# 10 · Capstone — Multi-Environment Platform

This capstone combines every Level 3 module: directory-per-environment
structure (module 06), remote state read across environments (Level 2
module 03 plus module 07's graph reasoning), state locking (module 01), a
policy check (module 05), and a `terraform test` suite (module 04) — built
around the Level 2 capstone's network/app modules. As with every module in
this course, this is reasoned through against documented behavior and was
not run against a real cloud account.

## Directory layout

```text
platform/
  modules/
    network/        (from Level 2 capstone)
    app/            (from Level 2 capstone)
  envs/
    dev/
      main.tf
      backend.tf
    prod/
      main.tf
      backend.tf
  policy/
    instance-type-allowlist.sentinel
  tests/
    app_monitoring.tftest.hcl
```

## `envs/prod/backend.tf` — isolated, locked remote state

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "platform/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-terraform-locks"
    encrypt        = true
  }
}
```

`envs/dev/backend.tf` is identical except for `key =
"platform/dev/terraform.tfstate"` — same bucket, same lock table, disjoint
keys, so a lock held for a `dev` apply can never block or interfere with a
`prod` apply, exactly module 01's locking model applied per-environment.

## `envs/prod/main.tf` — composing the Level 2 modules per-environment

```hcl
locals {
  environment = "prod"
  name_prefix = "acme-${local.environment}"
}

module "network" {
  source     = "../../modules/network"
  name       = local.name_prefix
  cidr_block = "10.2.0.0/16"
  az_count   = 3
}

module "app" {
  source            = "../../modules/app"
  name              = local.name_prefix
  vpc_id            = module.network.vpc_id
  subnet_ids        = module.network.public_subnet_ids
  instance_count    = 4
  enable_monitoring = true
}

output "vpc_id"           { value = module.network.vpc_id }
output "app_instance_ids" { value = module.app.instance_ids }
```

`envs/dev/main.tf` calls the identical modules with `az_count = 2`,
`instance_count = 1`, `enable_monitoring = false` — the same
directory-per-environment structure from module 06, now with each
environment's state independently locked and encrypted.

## `policy/instance-type-allowlist.sentinel` — enforced on every environment's plan

```hcl
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium", "t3.large"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type != "aws_instance" or
    rc.change.after.instance_type in allowed_types
  }
}
```

Attached (in a real Terraform Cloud organization) as `hard-mandatory` to
both the `dev` and `prod` workspaces — the same policy governs both
environments, since the allowlist is an organizational rule, not an
environment-specific one, directly reusing module 05's pattern.

## `tests/app_monitoring.tftest.hcl` — regression-testing the conditional

```hcl
mock_provider "aws" {}

run "dev_has_no_monitoring" {
  command = plan
  module {
    source = "../modules/app"
  }
  variables {
    name              = "acme-dev"
    vpc_id            = "vpc-mock"
    subnet_ids        = ["subnet-mock"]
    instance_count    = 1
    enable_monitoring = false
  }
  assert {
    condition     = length(aws_cloudwatch_metric_alarm.high_cpu) == 0
    error_message = "Dev should not create a monitoring alarm"
  }
}

run "prod_has_monitoring" {
  command = plan
  module {
    source = "../modules/app"
  }
  variables {
    name              = "acme-prod"
    vpc_id            = "vpc-mock"
    subnet_ids        = ["subnet-mock"]
    instance_count    = 4
    enable_monitoring = true
  }
  assert {
    condition     = length(aws_cloudwatch_metric_alarm.high_cpu) == 1
    error_message = "Prod should create a monitoring alarm"
  }
}
```

Testing `modules/app` directly (via the `module { source = ... }` block
inside the test file) rather than a whole environment means this test
suite runs in seconds against a mock provider and catches a regression in
the monitoring conditional regardless of which environment's `main.tf`
would have exercised it.

## The full promotion workflow

```bash
cd platform
terraform test                        # fast, no cloud creds needed

cd envs/dev
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
conftest test plan.json               # or: policy check runs automatically in TFC
terraform apply tfplan

# ... validate dev ...

cd ../prod
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
conftest test plan.json
terraform apply tfplan
```

## How It Actually Works

**Every mechanism this capstone combines operates at a different stage of
the same underlying plan/apply pipeline, which is exactly why they compose
without interfering with each other.** State locking (module 01) guards
the read-plan-apply window; policy evaluation (module 05) inspects the
completed plan's JSON before any `ApplyResourceChange` RPC fires;
`terraform test` (module 04) runs the identical graph-walk machinery
against an isolated, throwaway state and a mock provider. None of these
share a code path with each other at the point they intervene, which is
why adding a policy check doesn't slow down or change what `terraform
test` verifies, and why per-environment state locking doesn't need any
special-casing to also work correctly under a policy-gated Terraform Cloud
workflow — each layer only ever sees the interface (a plan, a lock
request, a mock RPC) appropriate to its own stage.

**Per-environment `backend` keys sharing one DynamoDB lock table produce
independent locks because the lock's identity is derived from the state
path, not the table** — module 01's compare-and-swap `PutItem` writes a
lock record keyed by `path` (the S3 object key, here
`platform/dev/terraform.tfstate` vs. `platform/prod/terraform.tfstate`),
so `dev` and `prod` applying concurrently acquire two entirely distinct
lock records in the same table with no contention between them at all —
sharing the table is purely an operational convenience (one resource to
provision and manage), not a source of cross-environment coupling.

**The `module { source = ... }` block inside a `.tftest.hcl` file causes
`terraform test` to build a graph rooted at that module directly, with the
test's own `variables` block standing in for what a calling root
configuration would normally supply** — this is precisely why the same
`modules/app` can be tested once, here, and reused unmodified by both
`envs/dev` and `envs/prod`: the test's mock inputs and each environment's
real inputs are just two different callers of the identical module
contract (inputs in, outputs out) established back in Level 2 module 01,
and neither caller has any way to observe which kind of caller the other
is.

## Exercise

Add a third environment, `envs/staging`, reusing the same `modules/network`
and `modules/app`, with `az_count = 2`, `instance_count = 2`, and
`enable_monitoring = true` — write its `backend.tf` (choosing an
appropriate `key`) and its `main.tf`'s `module "app"` block, then state
which of the two existing `tests/app_monitoring.tftest.hcl` `run` blocks
would need a new sibling to specifically cover staging's monitoring
behavior, if any.
