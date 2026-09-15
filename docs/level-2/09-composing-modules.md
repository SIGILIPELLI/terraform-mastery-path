---
description: "Composing Modules — Real infrastructure is rarely one module — it's several, wired together: a networking module producing a VPC ID, an app module…"
---

# 09 · Composing Modules

Real infrastructure is rarely one module — it's several, wired together:
a networking module producing a VPC ID, an app module consuming it, a
database module consuming both. This module covers passing values between
sibling modules and the ordering guarantees Terraform provides for free.

## Module outputs feeding module inputs

```hcl
module "network" {
  source = "./modules/network"
  cidr   = "10.0.0.0/16"
}

module "app" {
  source    = "./modules/app"
  vpc_id    = module.network.vpc_id
  subnet_id = module.network.public_subnet_id
}

module "database" {
  source    = "./modules/database"
  vpc_id    = module.network.vpc_id
  subnet_id = module.network.private_subnet_id
}
```

`module.network.vpc_id` is exactly the same reference syntax as reading a
resource's attribute — a module call is addressed as `module.<name>`, and
its declared `output` blocks are its attributes from the outside. Both
`app` and `database` depend on `network` through this reference; neither
depends on the other, so Terraform is free to create (or destroy) them in
any order, or concurrently, once `network` is satisfied.

## Passing a whole object between modules

```hcl
# modules/network/outputs.tf
output "vpc" {
  value = {
    id                = aws_vpc.this.id
    public_subnet_id  = aws_subnet.public.id
    private_subnet_id = aws_subnet.private.id
  }
}
```

```hcl
module "app" {
  source = "./modules/app"
  vpc    = module.network.vpc
}
```

```hcl
# modules/app/variables.tf
variable "vpc" {
  type = object({
    id                = string
    public_subnet_id  = string
    private_subnet_id = string
  })
}
```

Bundling related values into one `object`-typed output/variable pair (as
opposed to three separate outputs) keeps the calling code shorter and the
module's "network shape" contract explicit in one `object(...)` type
constraint that documents exactly what a caller can rely on.

## Explicit ordering with `depends_on` on a module

```hcl
module "iam_role" {
  source = "./modules/iam-role"
}

module "app" {
  source     = "./modules/app"
  depends_on = [module.iam_role]
}
```

Used when `app` needs `iam_role` to exist first for reasons Terraform
can't infer from a data reference — e.g., `app`'s resources rely on an IAM
policy attachment that isn't itself piped through an output/input pair
(a common case: eventual-consistency behavior in the underlying cloud API
that has nothing to do with Terraform's own dependency tracking). Like
resource-level `depends_on`, this is a deliberate escape hatch, not the
default way to establish ordering — an implicit reference (module input
using module output) is preferred whenever the dependency is expressible
that way, because it self-documents *why* the ordering exists.

## Worked example: three modules, one root

```hcl
module "network" {
  source = "./modules/network"
  cidr   = "10.0.0.0/16"
}

module "database" {
  source    = "./modules/database"
  vpc_id    = module.network.vpc_id
  subnet_id = module.network.private_subnet_id
}

module "app" {
  source      = "./modules/app"
  vpc_id      = module.network.vpc_id
  subnet_id   = module.network.public_subnet_id
  db_endpoint = module.database.endpoint
}
```

The reference chain alone establishes the full build order: `network`
first (nothing depends on anything), then `database` (needs `network`'s
outputs) can run concurrently with anything else only depending on
`network`, and `app` last (needs both `network`'s and `database`'s
outputs) — with zero explicit `depends_on` anywhere, because every real
dependency is expressed as a value reference.

## How It Actually Works

**Module composition doesn't create a new kind of graph edge — it's the
identical "reference creates dependency" rule from Level 1, just crossing
a module-call boundary.** When Terraform Core builds the configuration
graph, `module.app`'s `vpc_id = module.network.vpc_id` argument is parsed
into a reference to the specific output node
`module.network.output.vpc_id`, which in turn depends on whatever resource
inside `network` computed that value (`aws_vpc.this.id`). The graph edge
that results connects `app`'s resources all the way back to `network`'s
`aws_vpc` resource, transiting through the two output/input "pass-through"
nodes — those nodes have no side effects of their own; they exist purely
to relay a value and to mark the crossing of a module boundary in the
graph.

**Because outputs are graph nodes, an output's own value is only "ready"
once every resource it depends on has actually applied — this is what
gives module composition its ordering guarantee without any manual
sequencing.** `module.database`'s `vpc_id = module.network.vpc_id`
argument cannot be evaluated to a concrete value until
`module.network.output.vpc_id`'s node has resolved, which cannot happen
until `aws_vpc.this`'s `ApplyResourceChange` has actually returned an ID.
Terraform's graph walker will not begin applying `database`'s resources
until that entire chain is satisfied — the ordering is a structural
consequence of data-flow, not a scheduling policy layered on top of it.

**`depends_on` on a `module` block adds a graph edge from *every* resource
inside the calling module to *every* resource inside the depended-on
module** — a much blunter instrument than a value reference, which only
creates an edge to the specific resource that produced the referenced
value. This is exactly why `depends_on` between modules should be reached
for only when no output/input reference can express the dependency: it
serializes far more of the graph than is usually necessary, which can
turn what would otherwise be safely-parallel resource creation into a
strictly sequential one.

## Exercise

Given a `modules/network` module producing outputs `vpc_id` and
`nat_gateway_ip`, and a `modules/app` module that needs both as inputs,
write the two `module` blocks (`network` and `app`) that wire them
together — using only value references, no `depends_on` — and state which
resource inside `network` `app`'s apply is ultimately, transitively
waiting on.
