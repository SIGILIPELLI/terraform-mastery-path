---
description: "Disaster Recovery for State & Infra — Level 3's module on refactoring state safely covered surgical, intentional state edits. Disaster recovery is the…"
---

# 08 · Disaster Recovery for State & Infra

Level 3's module on refactoring state safely covered surgical,
intentional state edits. Disaster recovery is the opposite scenario:
something has already gone wrong — a state file was deleted, corrupted,
or an entire region went dark — and the question is how to get back to a
known-good, working configuration with the least additional damage.

## The state file is the single point of failure state operations create

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tfstate"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-tf-locks"
  }
}
```

Lose `terraform.tfstate` — accidentally delete the S3 object, corrupt it
with a bad manual edit, or have the bucket itself destroyed — and
Terraform loses all memory of which real-world resources it's supposed
to be managing. The infrastructure keeps running (state loss doesn't
touch actual cloud resources), but the *next* `terraform plan` sees an
empty state against a populated configuration and proposes creating
everything from scratch, which would either fail loudly (most cloud
resources reject creating something that already exists, like an S3
bucket name collision) or, worse, some resources *would* succeed and
silently duplicate infrastructure. This is why state is the actual
disaster-recovery-critical asset in a Terraform-managed environment —
not the `.tf` files, which are just text in version control and trivially
recoverable.

## Line of defense one: versioned, replicated state storage

```hcl
resource "aws_s3_bucket" "tfstate" {
  bucket = "acme-tfstate"
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_replication_configuration" "tfstate" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.tfstate.id

  rule {
    id     = "replicate-to-dr-region"
    status = "Enabled"
    destination {
      bucket = "arn:aws:s3:::acme-tfstate-dr-west"
    }
  }
}
```

Versioning on the state bucket means an accidental delete or a bad
overwrite is recoverable — S3 keeps every prior version of the object,
so restoring is "promote the previous version," not "reconstruct from
scratch." Cross-region replication protects against losing the entire
bucket or region, not just one object version. Neither of these is
optional hardening for a production Terraform setup; they're the
baseline that makes every other recovery step in this module possible at
all, because every one of them assumes *some* prior good state exists
somewhere to recover to.

## Recovering a lost or corrupted state file

```bash
aws s3api list-object-versions --bucket acme-tfstate --prefix production/terraform.tfstate
aws s3api get-object --bucket acme-tfstate --key production/terraform.tfstate \
  --version-id abc123def456 recovered.tfstate

terraform state push recovered.tfstate
```

`terraform state push` uploads a local state file as the new remote
state, after Terraform checks it against the backend's serial/lineage
metadata to avoid silently clobbering a newer concurrent write. Recovery
is: identify the last known-good version (via S3 versioning, or a backup
taken by CI before each apply), pull it down, verify it against reality
with `terraform plan` (expect *zero* changes if the recovered state
genuinely matches deployed infrastructure), and only then push it back
as the authoritative state.

## When there's no backup at all: reconstructing state from real infrastructure

```bash
terraform import aws_vpc.this vpc-0abc123
terraform import aws_subnet.main subnet-0def456
terraform import aws_db_instance.primary acme-prod-db
```

If no state backup exists, the only path back is importing every
existing resource one at a time — `terraform import` binds a real
resource ID to a resource address already declared in configuration,
populating that one resource's state without touching anything else.
This is slow and error-prone at scale (dozens or hundreds of resources,
each needing the exact right address and ID), which is precisely the
argument for why versioned/replicated state storage isn't optional: the
alternative is a manual, resource-by-resource reconstruction under
incident pressure, with real risk of missing a resource entirely and
having Terraform propose destroying something still in production use.

## Worked example: recovering from a region outage with a warm-standby workspace

```hcl
# dr/main.tf — separate state, separate workspace, deployed to us-west-2
terraform {
  backend "s3" {
    bucket = "acme-tfstate"
    key    = "production-dr/terraform.tfstate"
    region = "us-west-2"
  }
}

module "app_stack" {
  source = "../modules/app-stack"
  region = "us-west-2"
  # sized down: standby capacity, scaled up only during failover
  instance_count = 1
}
```

```bash
# Failover: scale up the DR workspace, then redirect traffic
terraform apply -var="instance_count=6" -target=module.app_stack
```

A genuinely resilient DR plan doesn't rely solely on state recovery — it
maintains a second, independently-stated deployment of the same
application stack in another region, kept minimally provisioned during
normal operation and scaled up on failover. This is architecturally
distinct from state recovery: it's accepting the primary region's *state
and infrastructure both* might become unavailable simultaneously, and
having a pre-tested, separately-applied configuration ready to take over
rather than depending on recovering anything from the failed region at
all.

## How It Actually Works

**S3 object versioning works at the storage layer, entirely below
Terraform's own state-locking and serial mechanisms — every `PUT` to the
state object creates a new immutable version with its own version ID,
regardless of what wrote it or whether that write went through
Terraform at all.** This is why it protects against failure modes
Terraform's own mechanisms don't cover: a locking bug or a bypassed lock
(someone running an old Terraform binary that doesn't support the
configured backend's lock protocol) can still produce a bad state write,
but it can never destroy the *previous* version, because versioning is
enforced by the storage backend, underneath and independent of anything
Terraform Core does.

**`terraform state push` isn't a raw overwrite — Terraform Core reads the
target backend's current state, compares its `serial` (a monotonically
incrementing counter embedded in every state file, bumped on every
write) and `lineage` (a UUID identifying which "family" of state history
a file belongs to, set once at state creation and never changed) against
the state being pushed, and refuses the push if the serial suggests it
would overwrite work newer than what you're pushing from.** This is the
exact mechanism (introduced in Level 3's refactoring-state module for
`state mv`) doing double duty here: the same lineage/serial check that
prevents two people's concurrent `apply`s from silently clobbering each
other also prevents a stale recovered backup from silently clobbering a
newer, valid state during a botched recovery attempt.

**`terraform import` populates exactly one resource's entry in state by
calling the provider's `ReadResource` RPC against the given real-world
ID and storing whatever attributes come back — it does not, and cannot,
infer *which* configuration block a given cloud resource should bind to,
which is why the resource address has to already exist in configuration
before you import into it.** This is a direct consequence of the
provider-protocol boundary covered across this course: state is
Terraform Core's local record of "what does resource address X
currently look like in the real world," and `import` is simply Core
asking the provider to answer that question for one specific ID and
recording the answer — it never runs the graph, never plans, never
touches any other resource, which is exactly why full reconstruction
requires one import call per resource with no bulk shortcut.

## Exercise

Your team's S3 state bucket has versioning enabled but no
cross-region replication, and an engineer just accidentally ran `aws s3
rb --force` against the bucket, deleting it and every version in it.
Walk through what recovery options remain, and explain — referencing the
lineage/serial mechanism above — what Terraform would do differently if
you instead tried to recover by writing a *freshly generated* state file
(from `terraform import`-ing everything) versus recovering an actual
prior version of the deleted file.
