---
description: "Designing a Platform's Terraform Architecture — Every module in this course so far has looked at one mechanism at a time — state locking, testing, policy…"
---

# 06 · Designing a Platform's Terraform Architecture

Every module in this course so far has looked at one mechanism at a
time — state locking, testing, policy, the graph. Designing a platform's
Terraform architecture is the opposite exercise: given all of those
mechanisms, how do you actually split dozens of teams' infrastructure
across state files, modules, and repositories so that nobody's `apply`
can take down someone else's service, and no one has to understand the
whole system to change their corner of it.

## The core tension: blast radius vs. duplication

```hcl
# One giant root module — everything in one state
module "network"  { source = "./modules/network" }
module "database"  { source = "./modules/database" }
module "app"       { source = "./modules/app" }
module "cdn"       { source = "./modules/cdn" }
```

```hcl
# vs. four separate root modules, four separate state files
# network/main.tf, database/main.tf, app/main.tf, cdn/main.tf
```

A single state file holding network, database, app, and CDN resources
means one `terraform apply` can, in principle, plan changes across all
four — which is convenient for keeping cross-references simple
(`module.network.vpc_id` is just a local reference) but means a lock held
by anyone touching *any* of it blocks everyone touching *any* of it (the
state-locking mechanism from Level 3 module 01), and a bad plan anywhere
in that state risks `-target`-less applies touching resources the
change had no business touching. Splitting into four state files
shrinks each blast radius to one domain, at the cost that
`module.network.vpc_id` is no longer a local reference — it has to cross
a state boundary via a remote state data source or an explicit output/
input contract, which is real coordination overhead. Platform
architecture is choosing where to draw these boundaries deliberately,
not accidentally.

## Composing across state boundaries: `terraform_remote_state`

```hcl
# database/main.tf
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "acme-tfstate"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_db_subnet_group" "main" {
  subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
}
```

The `network` state file's `outputs` block is the *only* thing the
`database` state file is allowed to depend on — not `network`'s internal
resource addresses, not its module structure, just whatever it chose to
publish as outputs. This is the actual API boundary between teams: the
network team can refactor everything inside their state (rename modules,
restructure subnets) as long as `private_subnet_ids` keeps meaning the
same thing, and the database team never needs to know or care how it's
produced. Get the output contract right and teams stop needing to
coordinate `apply` order by hand.

## The alternative to remote-state coupling: a platform module registry

```hcl
module "vpc" {
  source  = "app.terraform.io/acme/vpc/aws"
  version = "~> 4.0"

  cidr_block = "10.20.0.0/16"
  az_count   = 3
}
```

Instead of every team's root module reaching into another team's state
via `terraform_remote_state`, a platform team can publish reviewed,
versioned modules (via a private registry — Terraform Cloud's or an
internal Git-tag-based one) that encode the org's approved patterns —
"this is what a compliant VPC looks like here." Consuming teams pin a
version and get updates on their own schedule by bumping it, rather than
being silently affected by another team's state changing underneath
them. This trades the remote-state model's tight, live coupling for a
looser, versioned one — the same tradeoff as pinning a library dependency
instead of building against another team's `HEAD`.

## Layering: foundational vs. workload state

```text
platform/
  foundation/        (state: foundation) — org, VPCs, IAM boundaries, DNS zones
  shared-services/   (state: shared-services) — logging, monitoring, artifact registry
  teams/
    checkout/        (state: checkout) — owned entirely by the checkout team
    catalog/         (state: catalog) — owned entirely by the catalog team
```

A workable large-org layout separates state by *rate and ownership of
change*, not just by service. `foundation` changes rarely and requires
the widest review (it underlies everything); `shared-services` changes
occasionally and is owned by a platform team; each team's own state
changes constantly and is owned entirely by that team. Each layer
consumes the layer below it via published outputs or registry modules,
never the reverse — `checkout`'s state can read `foundation`'s VPC ID,
but `foundation`'s configuration never references anything about
`checkout`. That one-directional dependency is what keeps a graph of
otherwise-independent teams from becoming a single, effectively
monolithic apply the moment enough cross-references accumulate.

## Worked example: onboarding a new service without touching foundation

```hcl
# teams/payments/main.tf
data "terraform_remote_state" "foundation" {
  backend = "s3"
  config  = { bucket = "acme-tfstate", key = "foundation/terraform.tfstate", region = "us-east-1" }
}

module "service" {
  source  = "app.terraform.io/acme/ecs-service/aws"
  version = "~> 2.1"

  vpc_id           = data.terraform_remote_state.foundation.outputs.vpc_id
  private_subnets  = data.terraform_remote_state.foundation.outputs.private_subnet_ids
  cluster_arn      = data.terraform_remote_state.foundation.outputs.ecs_cluster_arn
  service_name     = "payments"
  container_image  = "acme/payments:1.4.0"
}
```

The payments team writes exactly this file, in their own state, applied
through their own CI pipeline, with their own approvers — no pull
request against `foundation`, no coordination with the platform team
beyond consuming the published `ecs-service` module and the foundation
outputs contract. This is the actual deliverable of good platform
architecture: a new team can stand up a compliant service by writing a
handful of lines against stable, versioned interfaces, without anyone
needing write access to, or deep knowledge of, any other team's state.

## How It Actually Works

**Each `terraform_remote_state` data source is a read of another
statefile's `outputs` map at plan time — Terraform Core fetches that
state file (via the same backend read path used for the owning
configuration's own state), extracts only the `outputs` block, and
exposes it as this data source's attributes.** Nothing about the
referenced state's resources, providers, or internal structure is
visible or loaded — the coupling is deliberately narrow, limited to
whatever the other configuration chose to declare as an `output` block.
This is mechanically why an output contract functions as a real API
boundary: Core enforces that only published outputs cross the boundary,
the same way a function's return value — not its local variables — is
the only thing its caller can observe.

**Splitting one state into several means the dependency graph that used
to be a single DAG inside one `terraform apply` becomes several
independent DAGs with no Core-level ordering between them at all** —
Terraform has no built-in concept of "apply `network`'s state before
`database`'s"; that ordering has to be enforced externally, by CI
pipeline sequencing or by the fact that `database`'s plan will simply
show stale or missing values if `network`'s outputs haven't been applied
yet. This is the direct cost of the reduced blast radius: you gain
independent locks and independent failure domains, and you give up
Core's automatic cross-resource ordering guarantee, which only ever
existed within a single state's single graph in the first place.

**A versioned registry module pins a specific module *source* at a
specific tag, so a consuming configuration's plan is computed against
exactly that version's resource definitions regardless of what the
module's source repository looks like today** — `terraform init`
resolves `version = "~> 4.0"` to a concrete tagged release and downloads
that snapshot, and nothing in the consuming state changes again until
someone bumps the version constraint and re-runs `init`. This is what
converts "another team's Terraform code" from a live, uncontrolled
dependency (remote state, which reflects whatever was last applied,
whenever that happened) into a controlled one (a module version,
which only changes when the consumer deliberately upgrades it) — the two
mechanisms in this module exist specifically to offer that choice.

## Exercise

A platform has one state file containing the org's VPC, its shared RDS
instance, and twelve independent microservices' ECS services, all
authored by different teams. Propose a state-splitting boundary (which
resources move to which state files, and how they'd reference each
other), and explain specifically how your split reduces the blast radius
of a mistake in one microservice's configuration compared to the
current single-state layout.
