# 01 · What Is IaC & Why Terraform?

Infrastructure as Code (IaC) means describing servers, networks, storage, and
other infrastructure in text files that a tool can read and turn into real
resources — instead of clicking through a cloud console or running one-off
scripts by hand.

## Why not just click through the console?

Clicking through a web console works, but it doesn't scale and doesn't leave
a trail:

- **No history.** Nobody can tell you *why* a setting changed or *who*
  changed it, short of digging through an audit log (if one even exists).
- **Not repeatable.** Rebuilding the same environment for staging, a second
  region, or disaster recovery means re-clicking every step, and small
  mistakes creep in.
- **Not reviewable.** A teammate can't look at a pull request before a change
  goes live — the change *is* the click.

IaC fixes this by making infrastructure changes look like software changes:
a diff, a review, a merge, then an apply.

## Declarative vs imperative

Terraform is **declarative**: you describe the *end state* you want, and
Terraform figures out the steps to get there.

```hcl
resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
}
```

This says "a bucket named `acme-reports-2026` should exist" — not "call the
create-bucket API, then set these five properties in this order." Contrast
that with an **imperative** approach (a shell script calling `aws s3
mb ...`, `aws s3api put-bucket-versioning ...`, one command at a time). An
imperative script has to be re-run carefully to avoid errors on a second run
("bucket already exists"); a declarative tool compares desired state to
actual state and only makes the difference happen.

## Where Terraform fits

Terraform (by HashiCorp) is a **provisioning** tool — it creates, updates,
and destroys the infrastructure resources themselves (compute instances,
networks, DNS records, IAM policies, managed databases, and so on), across
many providers (AWS, Azure, GCP, Kubernetes, GitHub, Datadog, and hundreds
more) through one consistent workflow and language.

It is commonly paired with, but distinct from:

| Tool category | Answers | Examples |
|---|---|---|
| Provisioning (Terraform) | "What infrastructure should exist?" | Terraform, Pulumi, CloudFormation |
| Configuration management | "What should be installed/configured *inside* a running machine?" | Ansible, Chef, Puppet |
| Container orchestration | "How should containers be scheduled and run?" | Kubernetes, Nomad |

A common pattern: Terraform provisions a VM and a Kubernetes cluster;
Ansible or a container image configures what runs inside the VM; Kubernetes
manifests describe what runs on the cluster.

## Why Terraform specifically

- **Provider-agnostic core.** The same workflow (`init`, `plan`, `apply`)
  works whether you're managing AWS resources, a GitHub organization, or a
  Datadog dashboard — you only need to learn a new provider's resource
  types, not a new tool.
- **Explicit plan before apply.** `terraform plan` shows exactly what will
  change *before* anything happens — reviewable in a pull request, like a
  code diff.
- **State-based reconciliation.** Terraform tracks what it created in a
  state file, so it knows the difference between "this doesn't exist yet"
  and "this exists and needs updating" without re-querying every possible
  resource on every run (module 07 covers this in depth).
- **Huge ecosystem.** The public Terraform Registry hosts thousands of
  providers and reusable modules maintained by HashiCorp, cloud vendors, and
  the community.

## A tiny end-to-end mental model

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
}
```

Reading this top to bottom: "Use the AWS provider, version 5.x. Talk to
`us-east-1`. Make sure a bucket called `acme-reports-2026` exists." Running
`terraform plan` against this shows "1 to add"; `terraform apply` creates it;
`terraform destroy` removes it. That four-word workflow —
**write, plan, apply, destroy** — is the shape of nearly everything you'll
do in this course, reasoned through against Terraform's documented behavior
rather than run against a live account in these lessons.

## Exercise

Without writing any HCL yet, list three infrastructure changes you've made
by hand recently (or imagine three: "created a storage bucket," "added a DNS
record," "opened a firewall port"). For each, write one sentence on what
could have gone wrong from doing it by hand that a reviewed, declarative
Terraform change would have caught (e.g., "a typo in the region wouldn't
have been reviewed by anyone before taking effect").
