---
description: "Terraform Cloud/Enterprise Concepts — Everything through Level 2 assumed you run terraform commands yourself, against a backend that's purely storage (S3…"
---

# 03 · Terraform Cloud/Enterprise Concepts

Everything through Level 2 assumed you run `terraform` commands yourself,
against a backend that's purely storage (S3, local disk). **Terraform
Cloud** (HCP Terraform) and **Terraform Enterprise** (the self-hosted
version) are a different kind of backend: one that also *runs*
`plan`/`apply` for you, on its own infrastructure, with locking, policy
enforcement, and a run history built in.

## Core concepts

- **Organization** — the top-level account boundary; contains workspaces.
- **Workspace (Terraform Cloud's, not the CLI's)** — despite the name
  overlap with module 04's `terraform workspace`, a Terraform Cloud
  *workspace* is a full unit of configuration + state + variables + run
  history, roughly equivalent to one whole configuration directory in the
  CLI world, not one slice of state within a single configuration.
- **Run** — one `plan`/`apply` cycle, executed remotely, with full logs
  retained and viewable by anyone with workspace access.
- **Variable sets** — variables shared across multiple workspaces (e.g. a
  shared cloud credential) without repeating them in every workspace.

## Configuring the `cloud` block

```hcl
terraform {
  cloud {
    organization = "acme-corp"

    workspaces {
      name = "reports-prod"
    }
  }
}
```

Replacing a `backend "s3" { ... }` block with `cloud { ... }` and running
`terraform login` (which stores an API token locally) switches state
storage, locking, *and* run execution to Terraform Cloud in one step —
running `terraform apply` locally after this no longer applies on your
own machine; it streams the plan to Terraform Cloud, which executes the
actual run remotely and streams logs back to your terminal.

## Remote runs: what changes vs. local execution

```bash
terraform plan
# Running plan in HCP Terraform. Output will stream here...
#
# Preparing the remote plan...
# Terraform v1.9.5
# ...
```

The plan is computed on Terraform Cloud's own runners, using variables and
credentials stored *there* rather than your local environment — meaning a
laptop with no cloud credentials configured at all can still trigger a
plan/apply, and every run is auditable centrally regardless of who
triggered it or from where.

## VCS-driven workflow

Terraform Cloud workspaces can be connected directly to a Git repository:
a push to the configured branch (or a pull request) automatically
triggers a `plan`, and merging can be configured to automatically trigger
the corresponding `apply` — the same trigger-on-push model Level 4's
CI/CD module builds by hand with GitHub Actions, but provided as a native
platform feature here.

## Sentinel / OPA policy enforcement (previewed; full depth in module 05)

```hcl
# a Sentinel policy attached to a workspace, evaluated between plan and apply
import "tfplan/v2" as tfplan

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type != "aws_instance" or
    rc.change.after.instance_type in ["t3.micro", "t3.small"]
  }
}
```

Terraform Cloud/Enterprise can run a **policy check** against every plan
before an apply is allowed to proceed — this specific policy would reject
any plan trying to create an EC2 instance larger than `t3.small`,
regardless of who requested the apply or whether they'd normally have
permission to run it. Module 05 covers policy-as-code mechanics in depth;
this is only how it plugs into the platform's run pipeline.

## Worked example: migrating an existing S3-backed configuration

```hcl
# before
terraform {
  backend "s3" {
    bucket = "acme-terraform-state"
    key    = "reports/terraform.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# after
terraform {
  cloud {
    organization = "acme-corp"
    workspaces {
      name = "reports-prod"
    }
  }
}
```

```bash
terraform login
terraform init
# Do you wish to proceed with importing the state you found?
#   Enter a value: yes
```

Identical state-migration prompt to Level 2 module 03's S3 example — from
`init`'s perspective, `cloud` is simply another backend implementation,
and switching to it goes through the exact same "detected backend change,
offer to migrate existing state" flow.

## How It Actually Works

**The `cloud` block isn't a variant of the ordinary state backend
interface — it fully replaces local execution with an API-driven remote
run, and the CLI becomes a thin client streaming input and output.** When
you run `terraform plan` with a `cloud` block configured, the local CLI
packages your configuration (and, depending on execution mode, either
uploads it or references the already-connected VCS revision), sends a
"create run" API call to Terraform Cloud, and then polls/streams that
run's log output back to your terminal in real time. The actual
`PlanResourceChange`/`ApplyResourceChange` provider RPC calls described in
Level 1 module 09 happen entirely on Terraform Cloud's own runner
infrastructure — your local machine's CPU, network, and credentials are
not involved in the plan/apply computation at all, only in initiating and
observing it.

**Terraform Cloud's workspace-level locking is enforced server-side as
part of the run queue, not via the DynamoDB-style mechanism module 01
described.** Because Terraform Cloud already owns run scheduling for a
workspace, it simply refuses to start a second run for the same workspace
while one is in progress — there's no separate lock object to acquire or
release because the run queue itself is the serialization point,
eliminating the local-lock-vs-crashed-process ambiguity module 01's
`force-unlock` exists to resolve (a stuck run can instead be manually
discarded through the platform's own UI/API, which is aware of exactly
which run is stuck and why).

**Policy checks are inserted as a distinct pipeline stage between plan and
apply, evaluated against the plan's *output*, not the `.tf` source.** The
Sentinel/OPA policy engine receives the same structured plan data
(`tfplan/v2` import above) that would otherwise just be rendered as
human-readable plan text — meaning a policy can inspect exactly what's
about to change (`rc.change.after.instance_type`) with full type
information, not by pattern-matching HCL source text, which is what lets
a single policy correctly govern every module and every way of writing an
`aws_instance` block that could produce a given planned change.

## Exercise

Rewrite the S3 `backend` block from Level 2 module 03's worked example as
a `cloud` block for an organization named `acme-corp` and a workspace
named `networking-prod`, and write the one CLI command (beyond `init`)
you'd need to run once, before `init`, to authenticate against Terraform
Cloud for the first time on a new machine.
