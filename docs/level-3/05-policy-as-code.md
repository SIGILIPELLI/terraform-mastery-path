---
description: "Policy as Code Concepts — Module 03 previewed Sentinel running between plan and apply on Terraform Cloud; this module goes deeper into policy as code…"
---

# 05 · Policy as Code Concepts

Module 03 previewed Sentinel running between plan and apply on Terraform
Cloud; this module goes deeper into **policy as code** generally — writing
organizational rules ("no public S3 buckets," "instances must be tagged
with a cost center") as version-controlled, automatically-enforced code
rather than a wiki page someone has to remember to check.

## The two dominant policy engines

- **Sentinel** — HashiCorp's own policy language, native to Terraform
  Cloud/Enterprise, evaluated directly against a run's plan data.
- **Open Policy Agent (OPA) / Rego** — a general-purpose, CNCF policy
  engine usable with Terraform via `conftest` against a JSON plan export,
  independent of any particular CI platform.

Both operate on the *same* underlying input: the structured JSON plan
(`terraform show -json`), meaning any policy either engine could enforce
is fundamentally a query over planned resource changes, not over your
`.tf` source text.

## A Sentinel policy: enforcing an instance-type allowlist

```hcl
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type != "aws_instance" or
    rc.change.after.instance_type in allowed_types
  }
}
```

Reads as: for every resource change in the plan, either it isn't an
`aws_instance` at all, or its planned `instance_type` is in the allowlist.
A plan proposing `t3.xlarge` fails this policy before `apply` is ever
permitted to run — regardless of who submitted it or whether they have
apply permissions on the workspace otherwise.

## The equivalent policy in Rego (OPA / conftest)

```rego
package terraform.instance_type

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_instance"
  allowed := {"t3.micro", "t3.small", "t3.medium"}
  not allowed[rc.change.after.instance_type]
  msg := sprintf("instance_type '%s' is not allowed for %s", [rc.change.after.instance_type, rc.address])
}
```

```bash
terraform show -json tfplan > plan.json
conftest test plan.json
# FAIL - plan.json - terraform.instance_type - instance_type 't3.xlarge' is not allowed for aws_instance.web
```

Same input document, same underlying logic, a different language and
execution model — `deny[msg]` accumulates violation messages rather than
returning a single boolean, which is idiomatic Rego but functionally
equivalent to Sentinel's `main = rule { ... }`.

## Advisory vs. hard-mandatory vs. soft-mandatory (Sentinel enforcement levels)

```hcl
policy "instance-type-allowlist" {
  source            = "./instance-type-allowlist.sentinel"
  enforcement_level = "soft-mandatory"
}
```

- `advisory` — failures are shown but never block the run.
- `soft-mandatory` — failures block the run, but an authorized user can
  explicitly override and proceed anyway (with the override recorded).
- `hard-mandatory` — failures block the run with **no** override path at
  all, for rules that must never be bypassed regardless of urgency
  (a compliance boundary, not a style preference).

Choosing the right enforcement level is itself a policy design decision:
too many `hard-mandatory` rules and legitimate emergency changes get
stuck; too many merely `advisory` and the rule set becomes decoration
nobody actually reads.

## Worked example: enforcing mandatory tags

```hcl
import "tfplan/v2" as tfplan

required_tags = ["Project", "Environment", "ManagedBy"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.change.after.tags is undefined or
    all required_tags as tag {
      tag in keys(rc.change.after.tags)
    }
  }
}
```

Directly enforces the `local.common_tags` convention Level 2 module 06
introduced by convention only — this policy makes it *mechanically
impossible* to merge a plan that skips one of the three required tag
keys, rather than relying on every contributor remembering the pattern.

## How It Actually Works

**Policy engines never see your `.tf` source — they consume the exact
same structured plan JSON that `terraform show -json` produces, meaning a
policy can only reason about what a plan actually *changes*, not about
style choices that never affect the plan's output.** `rc.change.after` in
both examples above is the provider's own planned-new-state representation
for that resource — the identical data Terraform Core received back from
`PlanResourceChange` in Level 1 module 09's graph walk, serialized to
JSON. This is precisely why a policy checking `instance_type` catches an
oversized instance no matter whether it was hardcoded, computed via a
`local`, or passed through three layers of module inputs (Level 2 modules
06 and 09) — by the time the plan exists, all of that indirection has
already been resolved down to one concrete planned value per resource.

**The plan-vs-apply pipeline position matters mechanically, not just
organizationally: policy evaluation happens after Terraform Core has
already computed the full plan but strictly before any `ApplyResourceChange`
RPC is issued.** This ordering is what makes policy-as-code a true
preventive control rather than a detective one — a `hard-mandatory`
Sentinel failure halts the run at the same point a human reviewer's
"request changes" would, with zero difference in blast radius, because no
provider RPC that would actually touch real infrastructure has fired yet.
Contrast this with Level 4's drift detection (module 03), which can only
ever be detective — it necessarily runs *after* infrastructure state has
already diverged, because there's no plan to intercept a change that
happened outside Terraform entirely.

**Sentinel's `import "tfplan/v2"` and Rego's `input.resource_changes` are
both, structurally, walking the identical JSON schema Terraform publishes
as its plan-representation format contract** — this is why the two
allowlist policies above are line-for-line translatable despite belonging
to different languages and platforms: `tfplan.resource_changes` and
`input.resource_changes` are the same array of the same objects, just
addressed through each language's own syntax for iterating and asserting
over structured data.

## Exercise

Extend the mandatory-tags Sentinel policy above so it also requires every
`aws_s3_bucket` resource change to have `versioning_configuration` present
(hint: check `rc.type == "aws_s3_bucket"` and inspect
`rc.change.after.versioning_configuration`), and state in one sentence
why this policy, unlike a code-review comment asking for the same thing,
cannot be silently forgotten in a rushed pull request.
