# 04 · Testing IaC — Plan Review

Terraform's built-in testing tools focus on catching problems **before**
`apply`, without needing real infrastructure — `validate` and `fmt -check`
at the syntax level, and, since Terraform 1.6, a real assertion-based
test framework (`terraform test`) that can run entirely against a mocked
or `plan`-only provider.

## `terraform validate`: the floor, not the ceiling

Already covered mechanically in Level 1 module 09 — it catches syntax and
type errors, nothing about whether the plan does what you actually intend.

## `terraform plan` as a review artifact

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
```

`-json` turns the plan into a structured, machine-readable document — this
is the artifact CI pipelines and policy engines (module 05) actually
inspect, rather than parsing the human-formatted terminal output. A code
review process built around Terraform commonly attaches the *human*
`terraform show tfplan` output to a pull request so reviewers see exactly
what will change before approving, closing the gap between "the diff
looks reasonable" and "I've actually seen what will happen to real
infrastructure."

## `terraform test`: assertion-based configuration tests

```hcl
# tests/bucket.tftest.hcl
variables {
  bucket_name = "test-bucket"
}

run "creates_bucket_with_correct_name" {
  command = plan

  assert {
    condition     = aws_s3_bucket.this.bucket == "test-bucket"
    error_message = "Bucket name did not match the input variable"
  }
}

run "versioning_disabled_by_default" {
  command = plan

  assert {
    condition     = length(aws_s3_bucket_versioning.this) == 0
    error_message = "Versioning resource should not exist when enable_versioning is false"
  }
}
```

```bash
terraform test
# tests/bucket.tftest.hcl... in progress
#   run "creates_bucket_with_correct_name"... pass
#   run "versioning_disabled_by_default"... pass
# tests/bucket.tftest.hcl... tearing down
# tests/bucket.tftest.hcl... pass
```

Each `run` block is one test case: `command = plan` runs only a plan
(fast, no real resources touched, ideal for asserting on configuration
logic like the conditional in module 07 of Level 2); `command = apply`
actually creates resources against a real provider and tears them down
afterward, useful for asserting on values a provider only computes at
apply-time, at the cost of real infrastructure being created during the
test run.

## Mocking providers for pure-logic tests

```hcl
mock_provider "aws" {}

run "instance_count_matches_input" {
  command = plan

  variables {
    instance_count = 3
  }

  assert {
    condition     = length(aws_instance.web) == 3
    error_message = "Expected 3 instances"
  }
}
```

`mock_provider` substitutes a fake provider that returns plausible
placeholder values for every attribute without ever calling a real API —
ideal for testing `count`/`for_each`/conditional logic (this course's
Level 2 topics) in complete isolation from cloud credentials, cost, or
network access, at the price of the mock's attribute values being
synthetic rather than what a real provider would actually return.

## Worked example: testing the Level 2 capstone's conditional monitoring

```hcl
# tests/app_monitoring.tftest.hcl
mock_provider "aws" {}

run "monitoring_disabled_in_dev" {
  command = plan
  variables {
    environment = "dev"
  }
  assert {
    condition     = length(aws_cloudwatch_metric_alarm.high_cpu) == 0
    error_message = "Dev should not create a monitoring alarm"
  }
}

run "monitoring_enabled_in_prod" {
  command = plan
  variables {
    environment = "prod"
  }
  assert {
    condition     = length(aws_cloudwatch_metric_alarm.high_cpu) == 1
    error_message = "Prod should create exactly one monitoring alarm"
  }
}
```

Two `run` blocks, each setting a different `environment` value, directly
verify the exact `count = var.enable_monitoring ? 1 : 0`-style logic from
Level 2 module 07 — a regression where someone accidentally flips that
conditional is caught by `terraform test` without ever touching AWS.

## How It Actually Works

**`terraform test` reuses the identical plan/apply graph-walking engine
this whole course has described — a `run` block is not a separate
interpreter, it's a full `plan` or `apply` invocation against a temporary,
isolated state, with assertions evaluated against the resulting plan or
state values afterward.** Each `run` block gets its own ephemeral state
(carried forward to the next `run` in the same file only if you
explicitly reuse it), so `assert` conditions are plain HCL boolean
expressions evaluated against real resource attribute values from that
run's plan — `aws_s3_bucket.this.bucket` in an assertion resolves exactly
the way it would in any other expression context, through the same
expression evaluator, just with the *source* of that value being a
completed plan rather than a `output` block.

**A mock provider intercepts at the exact RPC boundary described back in
Level 1 module 07 and 09** — `ReadResource`/`PlanResourceChange`/
`ApplyResourceChange` calls that would normally go to the real `aws`
plugin binary are instead answered by Terraform's own mock-provider logic,
which reads each resource type's schema (the same schema the real
provider publishes) and synthesizes a value of the correct type for every
computed attribute it doesn't have a concrete answer for. This is why mock
tests can assert on *counts* and *structural* facts (does this resource
exist, is this argument what I set it to) reliably, but can't meaningfully
assert on values a real provider computes (an actual AWS-assigned ARN) —
the mock has no real API to ask, so those attributes are synthetic
placeholders, not predictions of real behavior.

**`command = apply` tests run the real graph against real providers and
real credentials, and Terraform automatically destroys everything the test
created at the end of the file** — a `run "..." { command = apply }` block
is tracked in the test's own ephemeral state exactly like a normal
`apply`, and `terraform test`'s teardown phase (visible in the CLI output
as "tearing down") walks that state in reverse-dependency order, the same
destroy-ordering rule from Level 1 module 09, specifically so a test suite
that creates real resources doesn't leave orphaned infrastructure behind
even if a later assertion in the same file fails.

## Exercise

Write a `.tftest.hcl` file with a `mock_provider "aws" {}` block and one
`run` block that sets `var.environment = "staging"` and asserts that
Level 2 capstone's `module.app` would be given `instance_count = 1` (i.e.
that staging is treated as non-prod) — you'll need one `run` block, one
`variables` block inside it, and one `assert` block.
