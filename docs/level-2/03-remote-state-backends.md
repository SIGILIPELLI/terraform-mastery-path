---
description: "Remote State & Backends — Level 1 module 07 flagged the two problems with local state: it isn't shared (a teammate running apply has no idea what your…"
---

# 03 · Remote State & Backends

Level 1 module 07 flagged the two problems with local state: it isn't
shared (a teammate running `apply` has no idea what your local
`terraform.tfstate` contains), and it isn't safe to commit (it can contain
plaintext secrets). A **backend** is what solves both — it tells Terraform
*where* to store state and, for backends that support it, how to lock it
during writes.

## What a backend actually configures

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "reports/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-terraform-locks"
    encrypt        = true
  }
}
```

- `bucket` / `key` — where in S3 the state file for *this* configuration
  lives. `key` is the path within the bucket, so many configurations can
  share one bucket by using different keys (`reports/terraform.tfstate`,
  `networking/terraform.tfstate`).
- `dynamodb_table` — a DynamoDB table used purely for **state locking**
  (covered mechanically in Level 3 module 01); S3 itself has no native
  locking primitive, so the AWS backend pairs it with DynamoDB to get one.
- `encrypt` — encrypts the state object at rest in S3.

The backend block cannot use variables or `locals` — its arguments must be
literal values (or supplied via `-backend-config` at `init` time), because
Terraform has to resolve *where the state lives* before it has evaluated
anything else in your configuration.

## Migrating from local to remote state

```bash
terraform init
# Initializing the backend...
# Do you want to copy existing state to the new backend?
#   Enter a value: yes
#
# Successfully configured the backend "s3"!
```

Adding a `backend` block to configuration that previously had none (or
changing one backend to another) and re-running `terraform init` triggers
a **state migration prompt** — Terraform detects the backend changed,
reads your current state from the old location, and offers to copy it to
the new one so no resource tracking is lost.

## Partial configuration with `-backend-config`

```hcl
terraform {
  backend "s3" {}
}
```

```bash
terraform init -backend-config="bucket=acme-terraform-state" \
                -backend-config="key=reports/terraform.tfstate" \
                -backend-config="region=us-east-1"
```

Leaving the backend block empty and supplying values at `init` time is how
CI pipelines and multi-environment setups (Level 3 module 06) avoid baking
environment-specific bucket names into version-controlled `.tf` files —
the same configuration is `init`'d against a different backend per
environment by passing different `-backend-config` flags or files.

## `terraform_remote_state`: reading another configuration's outputs

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "acme-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_id
  # ...
}
```

This is how separately-managed configurations (a networking team's state,
a per-environment state — see Level 3 module 06) share values without
being merged into one giant configuration: the *consumer* declares a
`terraform_remote_state` data source pointing at the *producer's* backend
and reads its `output` values, read-only.

## Worked example: two configurations sharing state via outputs

```hcl
# networking/outputs.tf (producer)
output "subnet_id" {
  value = aws_subnet.main.id
}
```

```hcl
# app/main.tf (consumer)
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "acme-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  subnet_id     = data.terraform_remote_state.network.outputs.subnet_id
}
```

The `app` configuration never touches the `networking` configuration's
resources or state file directly — it can only see whatever `networking`
deliberately exposed via `output`, which is the same input/output boundary
module 01 introduced, just enforced across configurations instead of
across a module call.

## How It Actually Works

**The backend is resolved and locked in before the configuration graph
exists.** `terraform init` reads the `terraform { backend "..." {} }`
block (plus any `-backend-config` overrides) and writes the resolved
settings into `.terraform/terraform.tfstate` (a small local file that
records *which* backend and config to use — not your infrastructure's
state itself). Every subsequent command in that directory reads this
pointer first, then asks the backend implementation to fetch the real
state. This two-step indirection is exactly why changing environments via
`-backend-config` requires re-running `init`: the pointer file, not just
the in-memory backend object, has to be rewritten.

**State migration is a byte-for-byte copy through the state-manager
interface, not a merge.** When `init` detects a backend change, it reads
the full state JSON from the *old* backend, writes it verbatim to the
*new* backend, and — only after that succeeds — updates the local pointer.
Terraform refuses to proceed if it can't confirm the write succeeded,
specifically to avoid the failure mode where a configuration ends up
pointing at a backend location that doesn't actually contain its history,
which would look identical to "nothing has ever been applied here" on the
next `plan`.

**`terraform_remote_state` is a plain data source, evaluated read-only,
with no locking implications for the reader.** Internally it calls the
exact same backend-read code path a normal `plan` uses to load *its own*
state, just pointed at someone else's backend config and exposing the
result's `outputs` map as the data source's attributes. Because it's a
read, the consumer never takes a lock on the producer's state — only the
producer's own `plan`/`apply` runs do that — which means a `terraform plan`
in `app/` can run concurrently with an `apply` in `networking/`, though it
may then be planning against outputs that are about to change underneath
it if the producer's apply completes mid-plan.

## Exercise

You have two configurations, `networking` (produces a `vpc_id` output) and
`database` (needs that `vpc_id` as an input). Write the `terraform_remote_state`
data source block `database` would need to read `networking`'s S3-backed
state, and the one-line reference expression `database` would use in an
`aws_db_subnet_group` resource to consume that `vpc_id`.
