---
description: "Capstone — Reusable Module Set — This capstone combines every Level 2 module into one small but complete system: a reusable networking module and a…"
---

# 10 · Capstone — Reusable Module Set

This capstone combines every Level 2 module into one small but complete
system: a reusable networking module and a reusable app module, composed
from a root configuration, with conditional resources, `for_each`-based
repetition, and shared naming/tagging via `locals`. As with every module
in this course, this was reasoned through against documented Terraform
and AWS provider behavior — it was not run against a real cloud account.

## Directory layout

```text
capstone/
  main.tf
  variables.tf
  outputs.tf
  modules/
    network/
      main.tf
      variables.tf
      outputs.tf
    app/
      main.tf
      variables.tf
      outputs.tf
```

## `modules/network` — reusable VPC + subnets

```hcl
# modules/network/variables.tf
variable "name"         { type = string }
variable "cidr_block"   { type = string }
variable "az_count"     {
  type    = number
  default = 2
}
```

```hcl
# modules/network/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = { Name = var.name }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  for_each          = toset(slice(data.aws_availability_zones.available.names, 0, var.az_count))
  vpc_id            = aws_vpc.this.id
  availability_zone = each.value
  cidr_block        = cidrsubnet(var.cidr_block, 4, index(data.aws_availability_zones.available.names, each.value))
  tags              = { Name = "${var.name}-public-${each.value}" }
}
```

```hcl
# modules/network/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}

output "public_subnet_ids" {
  value = [for s in aws_subnet.public : s.id]
}
```

`cidrsubnet(var.cidr_block, 4, N)` carves the module's own CIDR block into
16 (`2^4`) equally-sized subnets and picks the `N`th one — a pure function
computed entirely by Terraform Core, needing no provider round-trip,
which is why it can safely run during `plan` even before the VPC itself
exists.

## `modules/app` — reusable app tier, with an optional feature

```hcl
# modules/app/variables.tf
variable "name"            { type = string }
variable "vpc_id"          { type = string }
variable "subnet_ids"      { type = list(string) }
variable "instance_count"  {
  type    = number
  default = 2
}
variable "enable_monitoring" {
  type    = bool
  default = false
}
```

```hcl
# modules/app/main.tf
resource "aws_instance" "this" {
  count         = var.instance_count
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  subnet_id     = var.subnet_ids[count.index % length(var.subnet_ids)]
  tags          = { Name = "${var.name}-${count.index}" }
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count               = var.enable_monitoring ? 1 : 0
  alarm_name          = "${var.name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods   = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}
```

```hcl
# modules/app/outputs.tf
output "instance_ids" {
  value = [for i in aws_instance.this : i.id]
}
```

`count.index % length(var.subnet_ids)` round-robins instances across
however many subnets the network module produced — module 08's `count`
pattern applied to a value (`var.subnet_ids`) that itself came from
another module's `for_each`-produced output.

## Root configuration — composing both modules

```hcl
# capstone/variables.tf
variable "project"     { type = string }
variable "environment" { type = string }

locals {
  name_prefix = "${var.project}-${var.environment}"
  is_prod     = var.environment == "prod"
}
```

```hcl
# capstone/main.tf
module "network" {
  source     = "./modules/network"
  name       = local.name_prefix
  cidr_block = "10.0.0.0/16"
  az_count   = local.is_prod ? 3 : 2
}

module "app" {
  source             = "./modules/app"
  name               = local.name_prefix
  vpc_id             = module.network.vpc_id
  subnet_ids         = module.network.public_subnet_ids
  instance_count     = local.is_prod ? 4 : 1
  enable_monitoring  = local.is_prod
}
```

```hcl
# capstone/outputs.tf
output "vpc_id" {
  value = module.network.vpc_id
}

output "app_instance_ids" {
  value = module.app.instance_ids
}
```

One `environment` variable — via `local.is_prod` — drives more
availability zones, more app instances, and monitoring alarms in prod,
while dev gets a smaller, cheaper, unmonitored footprint from the exact
same two modules.

## How It Actually Works

**This capstone's graph spans three module scopes, but Terraform Core
still builds and walks exactly one graph — the module boundaries described
in modules 01 and 09 are a namespacing and evaluation-scope concept, not a
separate execution unit.** `module.app`'s `subnet_ids = module.network.public_subnet_ids`
argument creates edges from every `aws_instance.this[*]` node inside
`app` back through `network`'s output node to the specific
`aws_subnet.public[*]` `for_each` instances that produced the list — a
single connected DAG that happens to visually nest by directory.

**The `for_each` → `count` handoff (network's subnet map feeding app's
`count.index`-based instances) is where the two repetition mechanisms'
different guarantees actually matter in practice.** If `var.az_count`
changes from 2 to 3, `network`'s `for_each` set gains one new key and adds
exactly one new subnet — the existing two are untouched. But `app`'s
`count`-based instances re-derive `subnet_id` via
`count.index % length(var.subnet_ids)`, and since `length(var.subnet_ids)`
just changed, *every* existing instance's `%` result can shift, which
Terraform will show as in-place updates (changing `subnet_id`) across
instances that conceptually didn't need to move — a direct, concrete
illustration of why module 08 recommends `for_each` over `count` whenever
elements have stable identity, here surfacing at the point where a
`count`-based consumer sits downstream of a `for_each`-based producer.

**`local.is_prod` fans out through the whole graph via ordinary value
references, so a single variable change is enough to alter three
independent parts of the plan (AZ count, instance count, and whether the
alarm resource exists at all) in one coherent `plan` run** — because all
three are downstream of the same `locals` node, Terraform's evaluator
computes `local.is_prod` once and every dependent recomputes from that one
resolved value, guaranteeing they can never disagree within a single plan.

## Exercise

Extend this capstone with a `modules/database` module (a single
`aws_db_subnet_group` resource, taking `subnet_ids` as an input and
exposing its `name` as an output), wire it into the root configuration
using `module.network.public_subnet_ids` as its input, and add a
`depends_on = [module.network]` to `module.app` even though `app` already
references `module.network`'s outputs directly — then explain in one
sentence why that `depends_on` would be redundant here.
