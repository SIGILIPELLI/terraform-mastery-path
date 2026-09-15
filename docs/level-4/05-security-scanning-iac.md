---
description: "Security Scanning for IaC — A misconfigured security group or a public S3 bucket is just as much a bug as a syntax error — the difference is terraform…"
---

# 05 · Security Scanning for IaC

A misconfigured security group or a public S3 bucket is just as much a
bug as a syntax error — the difference is `terraform validate` has no
opinion on it, because "world-readable storage bucket" is perfectly
valid HCL. This module covers static analysis tools purpose-built to
catch that category of defect before `apply`, and — more importantly —
how they actually reason about your configuration to do it.

## The gap `terraform validate` leaves open

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "company-customer-exports"
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

`terraform validate` checks that every argument is the right type, every
required argument is present, and every reference resolves — by every one
of those criteria, this configuration is completely valid. It is also a
bucket with all four public-access protections explicitly disabled.
Schema validation and security posture are orthogonal concerns: the
provider schema for `aws_s3_bucket_public_access_block` declares
`block_public_acls` as a plain boolean with no notion that `false` is a
dangerous value for a "customer exports" bucket — that judgment requires
a rule written by someone who understands the *service's* security
model, not the *schema's* type model.

## Static analysis: pattern-matching against known-bad shapes

```bash
tfsec .
```

```text
Result #1 HIGH Public access block is disabled
────────────────────────────────────────────
  aws_s3_bucket_public_access_block.data
    [restrict_public_buckets] is set to false

  Impact  Public buckets are accessible to anyone
  Resolution  Set restrict_public_buckets to true
```

Tools like tfsec (now folded into Trivy) and Checkov work by parsing HCL
into the same syntax tree Terraform's own parser produces, then walking
it against a library of rules — each rule is a small predicate over
resource type and attribute values ("if `aws_s3_bucket_public_access_block`
exists and any of these four attributes is `false`, flag it"). No cloud
API call happens; no provider is even initialized. This is what makes
static IaC scanning fast enough to run on every commit — it's pure
syntax-tree pattern matching against a rule library, closer to a linter
than to the plan/apply pipeline.

## Custom rules with Open Policy Agent / Rego

```rego
package terraform.security

deny[msg] {
  resource := input.resource.aws_security_group[name]
  rule := resource.ingress[_]
  rule.cidr_blocks[_] == "0.0.0.0/0"
  rule.from_port == 22
  msg := sprintf("aws_security_group.%s allows SSH from anywhere", [name])
}
```

```hcl
resource "aws_security_group" "web" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Where tfsec/Checkov ship a fixed rule library, Conftest running Rego
policies (or Sentinel, covered for governance in module 07) lets an
organization encode its *own* judgment calls — "no SSH from
`0.0.0.0/0`" isn't a universal security truth (a bastion host might need
exactly this), it's a policy this specific org has decided to enforce.
Conftest evaluates Rego rules against either the raw HCL (via an
HCL-to-JSON conversion) or, more powerfully, against the JSON plan output
from module 04 — the latter lets a rule reason about the fully resolved
`after` state, including values that only exist post-interpolation, not
just the literal source text.

## Scanning the plan vs. scanning the source

```bash
terraform show -json tfplan > plan.json
conftest test --policy security-policies/ plan.json
```

Scanning raw `.tf` source (tfsec's default mode) catches problems visible
in the literal configuration, but misses anything that only resolves at
plan time — a `cidr_blocks` value built from a variable, a module output,
or a `for_each` expansion. Scanning the JSON plan instead means the rule
evaluates against `resource_changes[].change.after`, which holds the
fully interpolated, fully expanded values Terraform actually computed —
every `for_each` instance materialized as its own entry, every variable
substituted. This is the same JSON plan artifact module 04's cost tooling
consumes, and for the identical reason: it's the one place a
downstream tool can see the *resolved* configuration without
re-implementing Terraform's own expression evaluator.

## Wiring scanning into the same review gate as tests and policy

```yaml
# .github/workflows/security-scan.yml
on: pull_request
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/tfsec-pr-commenter-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
      - run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > plan.json
          conftest test --policy security-policies/ plan.json
```

This sits in exactly the same CI stage as the plan-review testing from
Level 3's module 04 and the policy-as-code checks from module 05 —
security scanning is one more static check against the plan artifact, run
before any human or automated approval gate, never after `apply`. Failing
it should block merge the same way a failed `terraform plan` or a failed
policy check does; catching a public bucket in code review costs nothing,
catching it after the bucket has held customer data for a week is an
incident.

## How It Actually Works

**A static IaC scanner never invokes the Terraform provider protocol —
it operates entirely on syntax trees or JSON, which is precisely why it
can evaluate configuration for a provider it doesn't have credentials
for, or even one it has never seen before.** tfsec and Checkov ship
their own HCL parsers (or reuse the open-source `hashicorp/hcl` library
directly) to build an abstract syntax tree independent of `terraform
init` ever running — no provider plugin is downloaded, no gRPC
connection is opened, no `GetSchema` call happens. The tool's rule
library encodes its own understanding of each provider's argument names
and dangerous values, maintained separately from — and sometimes lagging
— the actual provider schema, which is the direct cause of both false
negatives (a new provider version adds an equivalently dangerous
argument the rule library doesn't know about yet) and false positives
(a rule assumes a default value that a provider version quietly changed).

**Scanning the JSON plan instead of raw HCL changes what's actually being
matched: `after` values are the *outputs* of Terraform's expression
evaluator, not the literal expressions written in the file** — a rule
matching `cidr_blocks[_] == "0.0.0.0/0"` against source HCL fails to
fire if the actual source reads `cidr_blocks = [var.allow_cidr]`, because
the literal string `"0.0.0.0/0"` never appears in the file; the same rule
against the JSON plan fires correctly because `change.after.cidr_blocks`
holds the resolved value after Terraform Core has already done variable
substitution during plan evaluation. This is the identical resolved-vs-literal
distinction module 04 draws for cost estimation — a downstream static tool
gets more accurate results the closer it consumes to Terraform's own
evaluated output, and less accurate results the more it re-parses
un-evaluated source.

**Rego's evaluation model (a declarative query language over structured
JSON) is why Conftest can express relationships an attribute-matching
rule library can't — a Rego rule can join across multiple resources in
the same plan** (e.g., "flag any `aws_security_group` referenced by an
`aws_instance` whose `ami` doesn't match an approved AMI list"), because
`input` in a Rego policy is the entire parsed JSON plan document, not one
resource in isolation — the policy author writes an arbitrary query over
that whole document, which is strictly more expressive than a scanner
whose rule format is fundamentally "match this attribute path against
this value."

## 🔀 Related lessons on other tracks

- [Kubernetes — 05 · Image Scanning & Supply Chain Security](https://sigilipelli.github.io/kubernetes-mastery-path/level-4/05-image-scanning-supply-chain/)

## Exercise

A security group resource's `cidr_blocks` argument is set to
`var.office_cidr`, and `office_cidr`'s default value in `variables.tf` is
`"0.0.0.0/0"` (left that way by accident). Explain why a tfsec scan of
the raw `.tf` source would likely miss this, and why a Conftest policy
run against `terraform show -json` output would catch it.
