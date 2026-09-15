---
description: "Conditional Expressions — HCL has no if/else statement — it has a single conditional expression, condition ? true_val : false_val, borrowed from C-like…"
---

# 07 · Conditional Expressions

HCL has no `if`/`else` statement — it has a single **conditional
expression**, `condition ? true_val : false_val`, borrowed from C-like
languages. Combined with `count` (previewed here, covered fully in module
08), it's how Terraform expresses "create this resource only if..." despite
having no imperative control flow at all.

## The ternary operator

```hcl
variable "environment" {
  type = string
}

locals {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
  is_prod       = var.environment == "prod"
}
```

Both branches must produce **compatible types** — `condition ? "a" : 5`
is a type error, because Terraform must know the result type of the
expression during validation, before it knows which branch will actually
be taken (that depends on `var.environment`'s runtime value, or in some
cases only becomes known during `apply`).

## Conditionally creating a resource: `count = condition ? 1 : 0`

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}
```

There is no `resource ... if ...` syntax — `count`'s numeric value is the
only lever, and `0`/`1` is how a boolean condition becomes "resource
doesn't exist" vs "resource exists exactly once." Referencing this
resource elsewhere requires indexing (`aws_cloudwatch_metric_alarm.high_cpu[0].arn`),
because as soon as a resource has a `count` argument — even one that
always evaluates to `0` or `1` — Terraform treats it as a list of
instances, not a single resource.

## Conditional with `try()` and `coalesce()`

```hcl
locals {
  # try(): fall back to the second expression if the first errors
  # (e.g. referencing a possibly-absent map key)
  vpc_id = try(var.network_config.vpc_id, "vpc-default")

  # coalesce(): first non-null argument
  bucket_name = coalesce(var.custom_bucket_name, "${local.name_prefix}-default")
}
```

`try()` and `coalesce()` aren't conditionals in the `? :` sense, but they
solve the same practical problem — "use this value, unless it isn't
available, in which case use this other one" — for cases where writing the
full ternary out (checking for null, or catching an evaluation error)
would be more verbose than the intent warrants.

## Worked example: environment-driven sizing and optional resources

```hcl
variable "environment"       { type = string }
variable "enable_backup"     {
  type    = bool
  default = true
}

locals {
  is_prod        = var.environment == "prod"
  instance_type  = local.is_prod ? "t3.large" : "t3.micro"
  backup_enabled = local.is_prod ? true : var.enable_backup
}

resource "aws_instance" "app" {
  ami           = "ami-0abcd1234"
  instance_type = local.instance_type
}

resource "aws_backup_plan" "app" {
  count = local.backup_enabled ? 1 : 0
  name  = "app-backup"
  # ... rule block omitted for brevity
}
```

Prod always gets backups regardless of the input variable (the ternary
inside `backup_enabled` overrides `var.enable_backup` when
`local.is_prod` is true); non-prod environments respect whatever the
caller passed for `enable_backup`. One `count` expression is the entire
mechanism for making a whole resource optional.

## How It Actually Works

**The ternary is evaluated during graph evaluation like any other
expression — both branches are parsed and type-checked, but only one is
ever actually computed per plan.** Terraform's expression evaluator
short-circuits at the value level: for `cond ? a : b`, it first resolves
`cond`, then evaluates *only* the selected branch. This matters when a
branch would otherwise error — `cond ? var.x : var.list[0]` is safe even
if `var.list` is empty, as long as `cond` is true, because the `false`
branch (`var.list[0]`, which would error on an empty list) is never
evaluated. `try()` builds on this same short-circuiting evaluator, just
generalized to catch evaluation errors from *any* number of candidate
expressions rather than a single boolean split.

**`count`'s conditional pattern works because `count` controls how many
graph nodes exist for a resource, decided during plan, before any
individual instance is planned.** When Terraform evaluates
`count = var.enable_monitoring ? 1 : 0`, it must fully resolve that
expression to a concrete number *before* it can expand
`aws_cloudwatch_metric_alarm.high_cpu` into the right number of resource
instance nodes in the graph — which is why `count` cannot depend on a
value that's only known after `apply` (an attribute from another resource
not yet created): the graph's shape itself would be undetermined until
partway through applying it, which Terraform's static-graph model doesn't
allow. This is the single most common `count`-related error message new
users hit, and it's a direct consequence of graph construction happening
before any resource is actually applied.

**Type unification across ternary branches happens at validation, using
HCL's type-conversion rules, not the runtime value.** `var.environment ==
"prod" ? "t3.large" : "t3.micro"` type-checks fine because both branches
are string literals; if one branch were a number and the other a string,
Terraform's type checker would attempt an implicit conversion (numbers
convert to strings automatically) rather than erroring immediately — the
type system is deliberately a bit permissive here, but a genuinely
incompatible pair (e.g., a string vs. a list) fails `terraform validate`
outright, before either branch's actual runtime value is ever computed.

## Exercise

Write a `count`-based conditional for an `aws_eip` (Elastic IP) resource
that's created only when `var.environment` is `"prod"` **or**
`var.force_eip` (a bool) is `true` — using a single boolean expression
combining `||`, referenced from one `count` argument.
