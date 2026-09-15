---
description: "Governance at Scale — Level 3's policy-as-code module introduced Sentinel/OPA as a way to enforce a single rule against a single plan. Governance at scale…"
---

# 07 · Governance at Scale

Level 3's policy-as-code module introduced Sentinel/OPA as a way to
enforce a single rule against a single plan. Governance at scale is the
organizational question sitting on top of that mechanism: across dozens
of teams and hundreds of state files, who is allowed to change what,
which rules apply to which workloads, and how do you enforce any of it
without a platform team manually reviewing every plan by hand.

## Why per-team IAM alone doesn't scale as governance

```hcl
resource "aws_iam_role" "checkout_ci" {
  name = "checkout-terraform-ci"
  assume_role_policy = data.aws_iam_policy_document.checkout_trust.json
}

resource "aws_iam_role_policy" "checkout_ci" {
  role   = aws_iam_role.checkout_ci.id
  policy = data.aws_iam_policy_document.checkout_permissions.json
}
```

Scoping each team's CI role to only the AWS actions and resource ARNs
their own state needs is necessary — the checkout team's pipeline
genuinely shouldn't be able to touch the payments team's DynamoDB tables
— but it's a *ceiling* on what's technically possible, not a statement of
what's *approved*. IAM can't express "no S3 bucket may be created without
encryption enabled" or "every new database must use an approved
instance-class tier" — those are judgments about configuration content,
which is exactly the gap policy-as-code fills, and exactly why governance
at scale needs both layers together: IAM bounds the blast radius of what
a team's pipeline *can* do, policy bounds what it's *allowed* to
configure within that.

## Centralizing policy: one policy set, many workspaces

```hcl
# Terraform Cloud policy set, attached to every workspace tagged "production"
policy "enforce-encryption" {
  enforcement_level = "hard-mandatory"
}

policy "require-tags" {
  enforcement_level = "soft-mandatory"
}
```

```python
# sentinel.hcl
policy "enforce-encryption" {
  source             = "./enforce-encryption.sentinel"
  enforcement_level  = "hard-mandatory"
}
```

A platform team writes and versions a policy set once, in its own
repository, and attaches it to every workspace matching a tag
(`production`, or a specific cost center) rather than every team copying
policy logic into their own pipelines. `hard-mandatory` fails the run
with no override; `soft-mandatory` can be overridden by someone with the
right permission, which is the mechanism for handling legitimate
exceptions without weakening the rule for everyone else. This is what
makes policy a *governance* tool rather than just a *testing* tool: the
policy set is owned and versioned centrally, and every workspace it's
attached to inherits changes to it automatically, the same way the
registry-module pattern from module 06 lets a platform team push updates
to consumers without touching consumers' own repositories.

## Scoping rules by workload sensitivity

```python
import "tfplan/v2" as tfplan

is_production = tfplan.variables.environment.value is "production"

encrypted_storage = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is not "aws_s3_bucket" or
        rc.change.after.server_side_encryption_configuration is not null
    }
}

main = rule {
    (not is_production) or encrypted_storage
}
```

Not every rule should apply uniformly — a sandbox account's throwaway
S3 bucket doesn't need the same encryption mandate as a production
customer-data bucket. A well-designed governance policy reads a signal
from the plan itself (a variable, a tag, a workspace name convention)
to decide whether it even applies, rather than a platform team
maintaining a separate policy set per environment. Encoding the
condition inside the rule — as above — keeps one policy file expressing
the actual governance intent ("production must be encrypted") rather
than scattering environment-specific copies across the org.

## Governance for structure, not just content: mandatory module sources

```python
import "tfplan/v2" as tfplan

approved_source_prefix = "app.terraform.io/acme/"

main = rule {
    all tfplan.module_calls as _, mc {
        mc.source matches approved_source_prefix
    }
}
```

Beyond checking resource attributes, a policy can check *where a
configuration's modules came from* — rejecting a plan that sources a
module from a random public GitHub repo instead of the org's reviewed
private registry. This is governance over the software supply chain of
the Terraform configuration itself: it doesn't matter how clean a
module's code looks if nobody vetted it, and this rule makes "only use
modules we've reviewed" a machine-enforced fact rather than a wiki page
nobody reads.

## Worked example: an exception workflow that doesn't erode the rule

```text
1. Team requests exception via ticket, citing specific resource and reason
2. Platform team reviews, approves for a bounded time window
3. `soft-mandatory` policy override applied to that specific run only,
   logged with requester, approver, and expiry
4. Exception tracked in a dashboard; expired exceptions re-trigger the policy
```

The failure mode of any org-wide policy is that a hard block on a
legitimate edge case gets "solved" by someone quietly weakening the rule
for everyone, permanently. A `soft-mandatory` override tied to a specific
run, with an expiry and an audit trail, keeps the exception scoped to
the one case that needed it — the policy stays strict for the next
hundred runs, and the platform team has a record of every time it was
relaxed and why, which is itself useful signal for whether the rule
needs to change.

## How It Actually Works

**A policy set attached at the workspace or organization level is
evaluated by the same Sentinel/OPA runtime described in Level 3's
policy-as-code module, but the *attachment* — which policies run against
which workspaces — is metadata Terraform Cloud/Enterprise manages
independently of any single configuration's code.** This is the
structural reason centralizing policy actually works as governance:
a team's own repository never contains the policy logic and therefore
cannot accidentally (or deliberately) delete or weaken it — the policy
lives in a separate, platform-team-owned repository, and the workspace's
tags or organization membership are what determine which rules apply,
not anything the workspace owner controls.

**`hard-mandatory` and `soft-mandatory` differ only in what happens on a
`false` result — a hard-mandatory failure halts the run unconditionally,
identically to a hard error, while a soft-mandatory failure halts the run
pending an explicit override recorded by a user with the platform's
"manage policy overrides" permission.** That permission is an IAM-style
authorization check inside Terraform Cloud/Enterprise itself, entirely
separate from the cloud-provider IAM roles governing what a CI pipeline
can call — which is exactly why the two layers in this module (provider
IAM and policy enforcement) are non-overlapping controls: one is
enforced by AWS/GCP/Azure at the API-call level, the other is enforced by
Terraform Cloud/Enterprise's own run pipeline before any API call is
ever attempted.

**A module-source policy like the one above works because
`tfplan.module_calls` is populated from the configuration's actual
module block source arguments as resolved during `terraform init` — the
resolved source string (registry address, Git URL, or local path) is
part of the plan's structured representation, not something the policy
has to re-parse from raw HCL.** This is the same principle from the
security-scanning module extended to configuration structure instead of
resource attributes: because the plan's JSON representation exposes
`module_calls` as a first-class array, a policy can govern *where code
comes from* using the identical rule-evaluation mechanism it uses to
govern *what values resources are configured with* — both are just
different fields on the same evaluated plan object.

## Exercise

An org wants: (1) every S3 bucket in any workspace tagged `production`
must have encryption enabled, no exceptions, and (2) every workspace,
production or not, must only source modules from the org's private
registry, but violations of rule 2 should be override-able for a
sandbox team experimenting with a new module. Write out which
enforcement level (`hard-mandatory` vs `soft-mandatory`) you'd assign to
each rule and justify the choice for each.
