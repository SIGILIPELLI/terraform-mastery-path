---
description: "Cost Estimation Concepts — Terraform will happily let you apply a plan that provisions a p4d.24xlarge fleet you meant to size as t3.micro. Nothing in…"
---

# 04 · Cost Estimation Concepts

Terraform will happily let you `apply` a plan that provisions a
`p4d.24xlarge` fleet you meant to size as `t3.micro`. Nothing in Core's
plan/apply cycle prices anything — it computes *what will exist*, not
*what it will cost*. This module covers how cost estimation actually
attaches to that workflow: where the pricing data comes from, what a
plan file gives a cost tool to work with, and the limits of estimating
cost from configuration alone.

## Why Terraform Core has no idea what anything costs

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
}
```

A `terraform plan` for this resource produces a diff: no instance → one
instance, with these exact attribute values. Provider schemas describe
*shape* (what arguments exist, what types they are), not *price* —
`instance_type` is just a string as far as the AWS provider's schema is
concerned. Pricing lives in a completely separate data source (each
cloud's billing/pricing API or a published price list) that no part of
Terraform Core, the AWS provider, or the plan format touches. A cost
estimation tool has to bring that pricing data itself and cross-reference
it against the plan.

## Reading a plan as structured data: `terraform show -json`

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
```

This is the mechanism every cost tool (Infracost, cloud-native
estimators, custom scripts) actually builds on. `-out=tfplan` saves the
plan in Terraform's binary plan format; `show -json` converts that into a
documented JSON schema containing `resource_changes` — one entry per
resource, with `change.after` holding every planned attribute value.

```json
{
  "resource_changes": [
    {
      "address": "aws_instance.app",
      "type": "aws_instance",
      "change": {
        "actions": ["create"],
        "after": {
          "instance_type": "t3.micro",
          "ami": "ami-0abcd1234",
          "root_block_device": [{ "volume_size": 8, "volume_type": "gp3" }]
        }
      }
    }
  ]
}
```

A cost estimator walks this array, and for every `resource_changes[].type`
it recognizes, extracts the pricing-relevant attributes
(`instance_type`, `volume_size`, region from the provider config) and
looks each one up against a price catalog. This is why cost tools need a
per-resource-type mapping table and go stale when a provider adds new
resource types or attributes faster than the tool's mapping is updated —
the JSON plan gives them everything, but only for types they know how to
read.

## What's estimable and what isn't

```hcl
resource "aws_lambda_function" "handler" {
  function_name = "process-events"
  runtime       = "python3.12"
  memory_size   = 512
  timeout       = 10
}
```

`aws_instance` cost is estimable almost exactly from configuration alone:
instance type and region map directly to an hourly rate. `aws_lambda_function`
cost is fundamentally different — pricing depends on invocation *count*
and *duration*, neither of which exists anywhere in the configuration or
the plan. A cost tool can price the *rate* (per-GB-second, per-request)
but has no way to project a dollar figure without an assumed or observed
usage volume. This is a structural limit, not a tooling gap: static
configuration analysis can price fixed-capacity resources
(compute, storage, provisioned throughput) reasonably well, and can only
show unit rates — never a total — for consumption-based ones, because the
consumption number simply isn't a property of the Terraform configuration.

## Worked example: pricing a plan diff with Infracost

```bash
infracost breakdown --path . --format json --out-file infracost.json
```

```hcl
resource "aws_db_instance" "primary" {
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  storage_type      = "gp3"
}
```

Infracost runs `terraform plan` (or reads an existing plan JSON) itself,
extracts `aws_db_instance.primary`'s `instance_class`,
`allocated_storage`, and `storage_type`, looks each up against a
maintained AWS pricing dataset scoped to the configured region, and
reports an estimated monthly cost broken into compute
(`db.t3.medium` hourly rate × 730 hours) and storage
(`allocated_storage` GB × `gp3` per-GB-month rate) line items. Run against
a plan that only *changes* `allocated_storage` from 100 to 200, it reports
the delta, not just the new total — which is the actual point of wiring
this into CI: reviewers see "+$14.60/month" on the pull request diff
instead of having to reconstruct that from the HCL by hand.

## Wiring it into review, not apply

```yaml
# .github/workflows/cost-estimate.yml
on: pull_request
jobs:
  cost:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: infracost/actions/setup@v3
      - run: infracost breakdown --path . --format json --out-file infracost.json
      - uses: infracost/actions/comment@v3
        with:
          path: infracost.json
          behavior: update
```

Cost estimation belongs at the same review stage as `terraform plan`
(covered in Level 3's testing module) — a pull-request comment showing
the dollar delta before merge — never as a gate that blocks `apply`
outright, since legitimate cost increases (scaling up for real load) are
common and the tool has no authority to judge whether a given increase is
justified. It's an information surface for the human reviewer, not a
policy engine; Level 4's governance module covers the distinct mechanism
(policy-as-code) for turning a numeric threshold into an actual block.

## How It Actually Works

**A cost tool never talks to Terraform Core's provider protocol or state
— it only ever consumes the JSON plan format (`terraform show -json`),
which is a stable, versioned, documented output contract deliberately
decoupled from Core's internals.** This is why cost tools can support
new Terraform versions without being rewritten each time: the JSON plan
schema is treated as a public interface with its own compatibility
promise, separate from whatever internal graph or RPC structures Core
uses to produce it. The tool's job reduces to two independent steps that
have nothing to do with Terraform's own execution model: parse
`resource_changes[].change.after` for known type/attribute pairs, and
join against an externally maintained price catalog it ships or fetches.

**The reason cost estimation cannot use `terraform.tfstate` instead of a
fresh plan is that state reflects the *last applied* configuration, and
a cost review needs to price the *proposed* one** — the JSON plan's
`change.after` block specifically represents "what the object's
attributes will be if this plan is applied," computed by evaluating the
current configuration against current provider schemas and (for
computed attributes with real dependencies) against real API responses
during the plan's refresh step. Pricing the diff between `before` and
`after` in that same JSON is mechanically what produces a delta rather
than a total — the tool is doing the same before/after comparison
Terraform Core does for the human-readable plan output, just against a
price catalog instead of against the resource schema.

**No cost tool can see quantities that only exist at runtime (invocation
counts, data-transfer bytes, request rates) because those values are not
resources, arguments, or outputs in the Terraform graph at all — they are
facts about production traffic that Terraform's declarative model has no
representation for.** This is the same category boundary the testing
module drew between what `terraform plan` can validate (shape, syntax,
policy against static values) and what it structurally cannot
(request-time behavior) — cost estimation inherits that exact boundary
because it's built entirely on the same static plan artifact.

## Exercise

A pull request adds `aws_instance.worker` with `instance_type =
"m5.xlarge"` and `count = 4`, and separately adds
`aws_lambda_function.processor` with `memory_size = 1024`. Explain, in
terms of what `terraform show -json` would contain for each resource,
why a cost tool can produce a specific monthly dollar estimate for the
first addition but only a per-GB-second unit rate for the second.
